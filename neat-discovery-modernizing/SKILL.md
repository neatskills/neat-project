---
name: neat-discovery-modernizing
description: Use when the objective is to modernize an existing codebase — produces a phased transformation roadmap grounded in a full system health assessment.
---

# Modernizing

**Role:** You are a software architect who scans an existing codebase and evaluates its health — producing a phased modernization roadmap.

## Overview

Structure of this run: setup → codebase scan → evidence collection → evaluation → phase derivation → pattern design → requirements → write → review (conditional) → retro.

## When to Use

Run when you have an existing codebase to modernize. Not for greenfield work, feature development, or incremental product delivery — use the discovery suite for those.

## Configuration

### Workspace

- Writes: `specs/discovery/modernization.md` (including its `## Legacy System Analysis` sub-sections: System Summary, Domain Summary, Module Map, Quality Findings, Improvements) and one `specs/discovery/requirements{NN}.md` per TP — all in Phase 7
- Never writes: `specs/system/**`, `specs/delivery/**`

**Artifact status:** Follow [../references/neat-project/artifact-status.md](../references/neat-project/artifact-status.md) for artifact status marker semantics.

**Schema checks:** After writing any artifact, verify it against the relevant schema in `references/modernizing-artifacts.md` — confirm all required keys are present.

### Definitions

- **TP-N ↔ requirements mapping:** each transformation phase (TP-N) in `modernization.md` produces one `specs/discovery/requirements{NN}.md` file. Phase 7 numbers them sequentially (TP-1 → requirements01.md, TP-2 → requirements02.md, TP-3a → requirements03.md, TP-3b → requirements04.md, etc.) so that `neat-discovery-solutioning`, `neat-delivery-planning`, and `neat-delivery-building` can execute each using their standard Iteration mechanisms — no special-case branches required in those skills.

## References

- [references/playbooks/dotnet.md](references/playbooks/dotnet.md) — .NET-specific scan targets, evaluation overrides, and phase-derivation rules; loaded in Phase 1 Step 3 when `.csproj`, `.sln`, `global.json`, or `packages.config` is detected.
- [references/playbooks/cobra.md](references/playbooks/cobra.md) — Cobra/Go CLI-specific scan targets, evaluation overrides, and phase-derivation rules; loaded in Phase 1 Step 3 when `go.mod` names `github.com/spf13/cobra`.

## Phase 1: Setup

Check for existing modernization run, establish legacy system path, and load tech-stack playbook if detected.

**Step 1 — Operating mode:** Check whether `specs/discovery/modernization.md` exists:

- **No** → new run. Proceed to Step 2.
- **Yes** → a prior run finished through Phase 7 (nothing partial to resume).

  > [1] Review (Recommended) — preserves completed Phase 1-7 work and continues from Phase 8
  > [2] Restart — use only if legacy analysis is incorrect or outdated; clears all `## Legacy System Analysis` sub-sections before proceeding to Step 2

  Restart never deletes `specs/system/` — those files are owned by solutioning and represent the target architecture, not the legacy scan. If `modernization.md` frontmatter contains `legacy_path`, offer to reuse it or change it.

**Step 2 — Legacy system path:** Ask for the path to the existing system to modernize:

> "What is the path to the existing (legacy) system? It will be scanned read-only. The current working directory is the new repo where output is written."

Store as `legacy_path`. Validate:

- Path must exist — block with error if absent.
- `legacy_path` must differ from the current working directory — block with error if equal: two-repo separation is required.

Record `legacy_path: {path}` in `modernization.md` frontmatter (written in Phase 7). Planning and building read this value to resolve legacy file references in task specs.

**Step 3 — Tech stack detection:** Scan `legacy_path` for file signatures:

| Tech stack | File signals |
| --- | --- |
| `dotnet` | `.csproj`, `.sln`, `global.json`, `packages.config` |
| `cobra` | `go.mod` containing `github.com/spf13/cobra` |

- Exactly one match → load `references/playbooks/{stack}.md` and announce:

  > Detected: {stack}. Loading {stack} playbook.

- Multiple stacks match (e.g. a polyglot repo) → load all matching playbooks — apply each playbook's overrides only to the modules/areas exhibiting that stack's file signals — and announce:

  > Detected: {stack-1}, {stack-2}. Loading both playbooks.

- Multiple stacks match but only some playbook files exist on disk → load the available ones, continue applying overrides only from the loaded playbooks, and announce:

  > Playbook for {missing-stack} not found — skipping.

- No match, or exactly one stack matched and its playbook file is absent → proceed without a playbook and announce:

  > No matching playbook — running default methodology.

The loaded playbook(s) persist as session state across all subsequent phases.

If Review mode (Phase 1 Step 1 [1]), proceed to Phase 8. Otherwise, proceed to Phase 2: Evidence.

## Phase 2: Evidence

Scan legacy codebase following codebase-analysis protocol, hold outputs in-memory for modernization.md sections, then collect supplementary documents from user to fill gaps.

**Step 1 — Codebase scan:** Scan `legacy_path` — Follow [../references/neat-project/codebase-analysis.md](../references/neat-project/codebase-analysis.md), Steps 1 and 4–9. Skip Step 2 (Contradiction Check — no PRD exists at this stage) and Step 3 (Staleness Check — that step inspects the *new* repo's own `specs/system/modules/`, unrelated to this read-only legacy scan; modernizing always performs a full scan of `legacy_path`). Do **not** call `codebase-analysis.md` directly — it has hardcoded writes to `specs/system/` and `specs/delivery/issues.md`. Run equivalent steps in-memory and hold all outputs as session state for the `modernization.md ## Legacy System Analysis` sub-sections (written in Phase 7):

- Steps 4, 6, 7 (modules, conventions, domain) → `### System Summary` (Architecture, Conventions/Patterns), `### Module Map`, `### Domain Summary`
- Step 5 (contracts) → `### System Summary § Contracts` — also include CLI commands exposed at each module's `entry_point`
- **Flows:** derive from workflow and orchestration patterns found during domain extraction (Step 7) — one entry per major end-to-end interaction discovered in the code (cross-module `internal_imports` call chains, `source_files` content) → `### System Summary § Flows`
- Step 8 (quality audit) → `### Quality Findings`; do **not** write to `specs/delivery/issues.md`
- Step 9 (operational tooling) → `### System Summary § Foundation`

Module schema: Follow [../neat-discovery-solutioning/references/modernizing-artifacts.md](references/modernizing-artifacts.md). Shared packages go to `#### Shared Packages` within `### Module Map`. Record frontmatter `codebase_analysis: ok` (written in Phase 7) when the scan completes. If a playbook is loaded and defines `## Evidence`: apply its additional scan steps and embed their output in the relevant sub-section.

**Step 2 — Document intake:** State what the codebase scan already answers for each evaluation dimension. Then ask the user for any documents that cover gaps:

> "Share any available documents — architecture diagrams, runbooks, ADRs, incident reports, DORA metrics, SLA contracts, compliance certifications, API consumer lists."

Only ask about genuine gaps the codebase scan could not answer.

Proceed to Phase 3: Evaluate.

## Phase 3: Evaluate

Score system health across dimensions using full evidence, run constraint checks, and hold Amber improvement tasks for Phase 5 assignment.

**Step 1 — Constraint Checks:** Follow [references/modernizing-methodology.md](references/modernizing-methodology.md) § Constraint Checks — evaluate first, as findings constrain pattern selection in Phase 5.

**Step 2 — Dimension Scoring:**

- Follow [references/modernizing-methodology.md](references/modernizing-methodology.md) § Dimension Scoring.
- Hold Amber dimension improvement tasks as session state for Phase 5 assignment.
- If a playbook is loaded and defines `## Evaluate`: apply its threshold and evidence-source overrides to the relevant dimensions in place of the standard signals.

Proceed to Phase 4: Derive Phases.

## Phase 4: Derive Phases

Determine which transformation phases (TP-N) are needed using derivation rules, then size and decompose any XL/XXL phases.

**Step 1 — Apply derivation rules:** If a playbook is loaded and defines `## Derivation`, apply its rules first, then Follow [references/modernizing-methodology.md](references/modernizing-methodology.md) § Phase Derivation Rules for any case not already matched. If no playbook is loaded: Follow § Phase Derivation Rules directly.

Record each phase's `Depends on` (TP-IDs) — Follow [references/modernizing-methodology.md](references/modernizing-methodology.md) § Phase Derivation Rules (Depends on).

**Step 2 — Size and decompose:** Size each resulting phase (S / M / L / XL / XXL) and decompose any XL or XXL phase until all phases are S / M / L. When decomposing, extend the phase ID with a lowercase letter suffix (e.g. TP-3 → TP-3a, TP-3b). Set TP-3b's `Depends on` = TP-3a; TP-3a's `Depends on` inherits TP-3's original `Depends on`. Phase 7 numbers sub-phases per Definitions § TP-N ↔ requirements mapping.

Proceed to Phase 5: Design.

## Phase 5: Design

Select transformation pattern for each TP, design technical approach with complexity/risk scoring, assign Amber improvements to relevant phases, and identify unassigned Ambers for Improvements table.

**Step 1 — Extend pattern catalogue:** If a playbook is loaded and defines `## Design`, extend the Pattern Selection table with its stack-specific patterns before Step 2.

**Step 2 — Design each phase:** For each transformation phase, Follow [references/modernizing-methodology.md](references/modernizing-methodology.md) § Pattern Selection and § Complexity & Risk Criteria to determine:

- **Pattern** — Follow § Pattern Selection
- **Architectural changes** — what changes in the system's container/component structure
- **Complexity** — Low / Medium / High, Follow § Complexity & Risk Criteria
- **Risk** — Low / Medium / High, Follow § Complexity & Risk Criteria
- **Improvement tasks** — check whether any Amber dimension from Phase 3 targets the same system area as this phase; if so, record its improvement task here. Once an Amber dimension's improvement task is assigned to a phase, mark it consumed and do not assign it again to any later phase.
- **Go/no-go criteria** — measurable exit condition; name specific tooling, metrics, and commands
- **Rollback** — concrete reversal steps

**Step 3 — Unassigned Amber check:** For any Amber dimension from Phase 3 never assigned to a phase (no phase touched its system area), record a row for `## Legacy System Analysis / ### Improvements` (written in Phase 7).

- This is the sole destination for unscheduled Amber rows (not `## Constraint Checks`).
- Use the row format `| {Dimension} | {improvement task description} |`. Planning's issue seeding targets this section, so it must stay machine-distinguishable from `## Constraint Checks`.

Proceed to Phase 6: Requirements.

## Phase 6: Requirements

Derive requirements from each TP's improvement tasks, draft as Requirements table rows with dependencies, present grouped by TP, and accept user adjustments.

**Step 1 — Extract candidates:** For each TP (in order), use the TP's `Improvement tasks` cell as the primary source — each listed task is a candidate requirement. If the cell is empty, draft 1–3 requirements from the TP Name, Pattern, and Triggered-by fields.

**Step 2 — Draft per candidate:**

- Draft each candidate as a `## Requirements` row — Follow [references/modernizing-artifacts.md](references/modernizing-artifacts.md).
- Warn and pause if any `Depends on` value has no match.
- Compare your output against the `## Requirements` schema in `references/modernizing-artifacts.md` — confirm all required keys are present.

**Step 3 — Present grouped by TP:** Show all derived requirements grouped under their parent TP. Ask the user to review, adjust text, add missing requirements, or remove any out of scope. Accept all changes before proceeding.

Proceed to Phase 7: Write.

## Phase 7: Write

Write modernization.md with Legacy System Analysis sections populated from Phase 2 in-memory outputs, then generate one requirements{NN}.md per TP, verify all schemas, and set status to final.

Create `specs/discovery/` if absent. Write `modernization.md` — Follow [references/modernizing-artifacts.md](references/modernizing-artifacts.md). Frontmatter must include `legacy_path: {path}` (set in Phase 1 Step 2) and `status: draft`. `modernization.md` must include the `## Legacy System Analysis` section with all five sub-sections: `### System Summary`, `### Domain Summary`, `### Module Map` (with `#### Shared Packages`), `### Quality Findings`, `### Improvements`. Compare your output against the `modernization.md` schema — confirm all required keys are present.

**Per-TP requirements files:** After writing `modernization.md`, produce one `specs/discovery/requirements{NN}.md` per transformation phase, numbered per Definitions § TP-N ↔ requirements mapping. Each file contains only the requirements whose `Source` column (from Phase 6) matches that TP. Follow [../neat-discovery-assessing/references/modernizing-artifacts.md](references/modernizing-artifacts.md) — Iteration `requirements{NN}.md` schema, `## Active` table. Each requirement inherits its parent TP's Size, Complexity, and Risk; Rationale = the TP's Go/no-go criteria. Omit `## Deferred`. Add a dependency map (Mermaid) derived from each requirement's `Requires` value (resolved from the TP's `Depends on`, per Phase 6 Step 2).

After completing all quality checks, set `modernization.md` to `status: final`.

Proceed to Phase 8: Review.

## Phase 8: Review

Review mode or post-write refinement - present artifact path, set to draft, conduct iterative revisions, and finalize when complete.

- Present the Markdown file path.
- Set the artifact to `status: draft` when beginning revisions.
- For each revision: re-read the relevant section of `modernization.md` first, then edit.
- Ask, repeat until done:

  > Anything else, or ready to move on?

- When review is complete, set `status: final`.

Proceed to Phase 9: Retro.

## Phase 9: Retro

Reflect on the modernizing process and generate retro output for skill improvement.

Follow [../references/neat-project/retro-template.md](../references/neat-project/retro-template.md) § Simple — `{skill-name}` = `neat-discovery-modernizing`.
