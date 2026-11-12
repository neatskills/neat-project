---
name: neat-discovery-solutioning
description: Use when assessing has completed — produces a Solution Design (initial), updates it for iterations, or provides revised review.
neatskills:
  requires:
    - neat-discovery-assessing
    - neat-discovery-modernizing
---

# Solutioning

**Role:** Solution architect — validates feasibility, identifies risks, designs the solution — produces `solution.md` (new or updated in place).

## Overview

Structure of this run: intake → mode routing → flow mapping → gap resolution → solution design → quality review → publish → review (conditional) → retro.

**Modes:**
- **Initial:** `prd.md` → flow mapping → gap resolution → solution drafting → quality review → `solution.md`
- **Iteration:** `requirementsNN.md` + build history → flows for new requirements → gap resolution → solution drafting → quality review → updated `solution.md` (in place)
- **Review:** finalized `solution.md` → Phase 7 (open discussion loop → revised output)
- **Modernization:** `modernization.md` → knowledge files from Legacy System Analysis → flow mapping → solution drafting → quality review → `solution.md`

## When to Use

Run when assessing has completed — design a solution (Initial), extend architecture for new requirements (Iteration), or review existing output. Not for: implementation-ready specs, framework/library selection, detailed API/schema design, infrastructure vendor selection — use `neat-delivery-planning`.

## Configuration

### Workspace

`<output-path>`: `./specs/discovery` — shared by both discovery skills.

| File | Operation |
| --- | --- |
| `<output-path>/solution.md` | `write` (create or update in place) |
| `specs/system/system.md` | `write` (initialize); read by `neat-delivery-planning` |
| `specs/system/modules/*.md`, `specs/system/modules/index.md` | `write` (brownfield codebase scan) |
| `<output-path>/prd.md`, `<output-path>/requirementsNN.md` | `read — do not write` |
| `specs/discovery/modernization.md` | `read` (Modernization mode, or baseline when `prd.md` is absent) |
| build history (path derived per Phase 1 Step 2) | `read` — Iteration mode only, tiered |
| `specs/delivery/issues.md` | `read` then `append` — brownfield codebase scan only (`codebase-analysis.md` Step 8) |

**Never writes:** `specs/delivery/**` other than `issues.md` as above.

**Artifact status:** Follow [../references/neat-project/artifact-status.md](../references/neat-project/artifact-status.md). **Schema checks:** After writing any artifact, verify against `references/solutioning-artifacts.md` schema — confirm all required keys present.

### Routing

Resolved during Phase 1 and held as session state. Subsequent phases reference these routes directly.

| Route | Options | Set in |
| --- | --- | --- |
| `{mode}` | `Initial` \| `Resume` \| `Iteration` \| `Review` \| `Modernization` | Phase 1 Step 1 |
| `{project-type}` | `greenfield` \| `brownfield` | Phase 1 Step 2 |

### Definitions

- **reflects_requirements:** frontmatter list in `solution.md` tracking which `requirementsNN.md` files have been incorporated (e.g., `reflects_requirements: [01, 02, 03]`)
- **unincorporated requirements:** any `requirementsNN.md` file with `status: final` whose NN is NOT in the `reflects_requirements` list — assessing has scoped it but solutioning hasn't yet incorporated it

## Quick Reference

**Mode detection shortcuts:**
- `modernization.md` exists + no `prd.md` + no final `solution.md` → Modernization
- `solution.md` absent → Initial
- `solution.md` status=draft → Resume
- `solution.md` status=final + unincorporated requirements → Iteration
- `solution.md` status=final + no unincorporated requirements → Review

**Architecture reference loading pattern:**
- Load all architecture references at Phase 4 Step 1 before writing begins
- Keep `app-design.md`, `infrastructure.md`, `agent-design.md` available until all sections referencing them are complete
- `integration.md` applies to Tech Stack and Integrations sections — unload after both written
- `code-structure.md` applies to Phase 4 Step 3 and Phase 5 Step 1 completeness check
- `solution-flows.md` for Phase 2 flow identification and Phase 5 Step 1 validation
- `solution-review.md` for Phase 5 Step 2 subagent prompt

**Frontmatter update pattern (solution.md):**
- Initial: `status: draft`, `reflects_requirements: []`
- Iteration: keep existing list, append NN from newly incorporated requirementsNN.md files
- Finalization: `status: final` (Phase 6)

**Schema validation:**
After writing any artifact, verify against `references/solutioning-artifacts.md` — confirm all required keys present

**Iteration mode regeneration:**
`solution.md` is full, self-contained document — regenerate all diagrams and sections to reflect complete architecture (prior + new elements together); keep "Owned containers" list cumulative across iterations

## Phase 1: Intake

Detect run mode (Modernization, Initial, Resume, Iteration, Review), determine project type, load discovery artifacts, and run codebase scan if needed to create system.md.

**Step 1 — Detect mode:** If `specs/discovery/modernization.md` exists and neither `prd.md` nor finalized `solution.md` (`status: final`) exist → `{mode} = Modernization` — error if `modernization.md` has no `## Requirements` section (re-run modernizing Phase 7). Once Modernization mode completes, treat `modernization.md` as `prd.md` substitute — Iteration requires `requirementsNN.md` files from modernizing Phase 7.

Otherwise check `<output-path>/prd.md` or `specs/discovery/modernization.md` — if neither exists, error and exit.

Check `<output-path>/solution.md`:

- **Does NOT exist** → `{mode} = Initial` — **Greenfield:** Verify `prd.md` exists with `status: final` (else error: "prd.md must be finalized before running solutioning"); **Brownfield:** Verify `requirements01.md` exists with `status: final` (else error: "No finalized requirements found. Run assessing first to create requirements01.md"); proceed to Phase 1 Step 2.
- **Exists with `status: draft`** → `{mode} = Resume` — resume at Phase 4 using existing draft `solution.md`; proceed to Phase 1 Step 2.
- **Exists with `status: final`** → check for unincorporated requirements: list all `requirementsNN.md` files in `<output-path>/`, read `solution.md` frontmatter `reflects_requirements: []` (default empty), compare — any `requirementsNN.md` with `status: final` whose NN is NOT in `reflects_requirements`? **Yes** → `{mode} = Iteration`; proceed to Phase 1 Step 2. **No** → `{mode} = Review`; skip Phase 1 Step 2–3 and Phase 2–6, proceed to Phase 7. If no `requirementsNN.md` files exist, offer Review only.

**Step 2 — Load:** Determine project type: if `specs/system/system.md` exists, read first line (`mode: greenfield` or `mode: brownfield`); otherwise use codebase indicators (manifests, source directories): codebase → brownfield, none → greenfield. Step 3 creates `system.md` when absent.

- **`{mode} = Modernization`:** Load from `modernization.md` instead of `prd.md` — `## Executive Summary` → context; `## Requirements` table → requirements; `## Constraint Checks` → constraints; `## Legacy System Analysis` sections → hold in-memory as current state (System Summary: Foundation, Architecture, Contracts, Flows, Conventions/Patterns; Domain Summary; Module Map; Quality Findings) — used in Step 3 to populate `specs/system/` files, do not read `specs/system/` directly; TP table Complexity and Risk columns → risk flags; NFRs: none (embedded in TP Go/no-go criteria); `{open:}` callout seeding (Phase 3) is skipped.

- **`{mode} = Initial` or `{mode} = Resume` (Greenfield):** Load `prd.md` — extract: Requirements and NFRs, Current State, Delivery Estimate (high-risk XL/XXL), Constraints and Integrations, Project Context (team structure, repository preference if multi-product, domain complexity + rationale, scale indicators).

- **`{mode} = Initial` or `{mode} = Resume` (Brownfield):** Load `specs/system/system.md` → current state baseline (Foundation, Architecture, Contracts, Domain Concepts, Business Rules) — if absent, load after Step 3 creates it; load `requirements01.md` → extract Active section (requirements and NFRs); load `prd.md` if exists (optional) → extract Project Context, Constraints, Integrations; if no prd.md: derive Project Context from system.md or use defaults (team structure = inferred, domain complexity = moderate).

- **`{mode} = Iteration`:** Load `specs/system/system.md` → current state baseline; load `prd.md` if exists (Greenfield always has it; Brownfield optional) → extract Project Context, Constraints, baseline requirements; if Brownfield with no prd.md: load `modernization.md` if present for baseline context, else proceed with system.md only; load `solution.md` (current architecture); load all unincorporated `requirementsNN.md` files (those with `status: final` whose NN is NOT in `solution.md`'s `reflects_requirements` list) → extract Active section from each; build history: if `specs/delivery/delivery.json` exists, derive path from `planning_model` — `phase-first` → `./specs/delivery/build-history.md`, `product-first` → `./specs/delivery/{product-name}/build-history.md` (ask which products if multiple) — load in tiers (most recent wave entries: all fields; older wave entries: Architecture Changes and Decisions only); proceed without it if absent.

**Step 3 — Codebase Scan:** Check system.md existence first — if `specs/system/system.md` exists: `{mode} = Iteration` → skip to operational tooling rescan below; other modes → proceed to Phase 2 (Resume: Phase 4, per Step 1). If `specs/system/system.md` does NOT exist: check for codebase indicators (manifests, source directories) — **Codebase exists:** auto-run code analysis to create system.md (read `specs/delivery/issues.md` entries for ISS-NNN ID continuity before running `codebase-analysis.md`; tell user "system.md not found. Running code analysis to create it..."; follow [../references/neat-project/codebase-analysis.md](../references/neat-project/codebase-analysis.md) Steps 1–9; write `mode: brownfield` to system.md; proceed to Phase 2); **No codebase:** greenfield project (initialize `system.md` with skeleton from [Artifacts](references/solutioning-artifacts.md); write `mode: greenfield` to system.md; proceed to Phase 2).

**Modernization track:** Before checking system.md — if `modernization.md` frontmatter records `codebase_analysis: ok` → populate knowledge files from in-memory `## Legacy System Analysis` instead of running filesystem scan (initialize system.md skeleton, populate from System Summary and Domain Summary; expand Module Map rows into individual module files with `scan_status: ok`; write shared-modules.md and index.md per `references/solutioning-artifacts.md`; write `mode: brownfield` to system.md; proceed to Phase 2). If `modernization.md` exists but `codebase_analysis: skipped` → warn "Modernization ran but codebase analysis was skipped — running it now", then continue with auto-run logic above.

**`{mode} = Iteration` — operational tooling rescan:** If `system.md` already exists (typical for Iteration mode) — check if `specs/system/modules/` directory exists and contains at least one module file — **If modules exist:** run [../references/neat-project/codebase-analysis.md](../references/neat-project/codebase-analysis.md) Step 9 only (updates `system.md § Foundation` operational tooling lines); **If no modules exist** (greenfield project that completed Initial building and now has code): run full codebase-analysis (Steps 1–9) to create initial module files; then proceed to Phase 2. Module data recorded in `specs/system/modules/*.md` during brownfield scan is read by Phase 4 Step 2 to populate L3 Component Diagrams and L4 Code Diagrams.

If `{mode} = Review`, proceed to Phase 7. Otherwise, proceed to Phase 2: Flow Mapping.

## Phase 2: Flow Mapping

Identify applicable flow types and document each with one-line name and Mermaid diagram, deriving from requirements (Initial) or new requirements plus existing architecture (Iteration).

**Step 1 — Flows:** Follow [references/solution-flows.md](references/solution-flows.md) to identify which flow types apply, then document flows for each required type. Open each flow with one-line **Flow Name** (e.g. `Flow 1: User Authentication (OAuth 2.0 + PKCE)`); group flows by type. **Initial:** derive flows from PRD's Requirements and NFRs. **Iteration:** map flows for new requirements (from unincorporated `requirementsNN.md` Active sections) using `solution.md` and build history as architectural context.

Proceed to Phase 3: Gap Resolution.

## Phase 3: Gap Resolution

Scan loaded artifacts for architectural ambiguities (runtime/language, operational decisions, API design, data storage, integrations, UX behavior, constraint conflicts), seed gap nodes, and resolve using gap-resolution protocol before solution design.

Scan loaded requirements, constraints, and flows for undecided architectural choices that would block solution design — do not limit to known patterns; any ambiguity forcing an assumption should be surfaced. Common categories (not exhaustive): **Backend runtime/language** — any component (API handler, worker, script, CLI) whose implementation language is unspecified (must be confirmed here, not deferred to planning; test framework and schema design both depend on it); **Operational decisions** — read operational tooling findings from `system.md § Foundation` (written Phase 1 Step 3) before evaluating, extend existing tools, introduce new tool only when finding is `none detected`, greenfield has no findings (treat every category as `none detected`), follow [references/architecture/infrastructure.md](references/architecture/infrastructure.md) (supplement its prescribed patterns, never contradict), seed decision nodes and reminders per `infrastructure.md` § Operational Decision Nodes; **API design decisions** — protocol (REST vs GraphQL vs WebSocket), authentication model, streaming vs request/response, pagination approach; **Data storage ambiguities** — any entity whose storage type or structure is unclear (including file-based and ephemeral data); **Integration unknowns** — any external system where protocol, auth mechanism, or data contract is unspecified; **UX behavior** — any interaction affecting architecture (streaming output, real-time updates, offline support); **Constraint conflicts** — any PRD constraint contradicting a natural architectural choice (flag and resolve before writing).

Then resolve gaps — Follow [../references/neat-project/gap-resolution.md](../references/neat-project/gap-resolution.md): seed per § Cross-Skill Continuity from `prd.md` and `solution.md` (Iteration: also unincorporated `requirementsNN.md` files), targeting `solution.md` — Modernization mode: skip this seeding (`modernization.md` does not embed `{open:}` callouts); `unresolved-outcome`: `open` for design choices, `risk` for gaps materially affecting sizing or feasibility; model any other genuine gap (classify per § Node Types); run frontier before Phase 4 — do not assume or defer.

Proceed to Phase 4: Solution Design.

## Phase 4: Solution Design

Generate Architecture diagrams (System Landscape, System Context, Container, Deployment, Component, Code), plan Code Structure, write remaining sections following architecture references, sync key sections to system.md.

**Architecture reference files:** [references/architecture/app-design.md](references/architecture/app-design.md) (components, data layer, architectural UX decisions), [references/architecture/infrastructure.md](references/architecture/infrastructure.md) (infrastructure and operational concerns), [references/architecture/agent-design.md](references/architecture/agent-design.md) (AI agent patterns — include when AI/agents in scope), [references/architecture/integration.md](references/architecture/integration.md) (external integration contracts), [references/architecture/code-structure.md](references/architecture/code-structure.md) (repository structure decision and generation rules). `app-design.md`, `infrastructure.md`, `agent-design.md` each apply to multiple sections — keep available until all sections that reference them are complete; `integration.md` applies to two sections (Tech Stack, Integrations) — keep available until both written, then do not re-read; `code-structure.md` applies to Step 3 here and Phase 5 Step 1 completeness check.

**PRD UX artifacts:** If loaded PRD (`prd.md`; Iteration: also unincorporated `requirementsNN.md` files) contains `## User Journeys` and `## Layout Design` sections, reference them when making frontend architecture decisions — example: "Per PRD § User Journeys, the booking flow is multi-step with branching → wizard interaction model" or "Per PRD § Layout Design, desktop uses 70/30 content/sidebar split → state management needs sidebar-aware routing". If technical constraint contradicts PRD UX (e.g., PRD shows single-page layout but SSO redirect requires multi-page navigation), embed `{open: topic}` callout in relevant solution.md section with conflict and recommended resolution — user resolves during Phase 7 (Review).

**Step 1 — Principles:** **High-level only:** decide pattern and category, defer library/version/implementation detail to planning; **Adopt over build:** default to proven open-source or SaaS, every build decision requires explicit rationale; **Keep it simple:** prefer simplest architecture meeting requirements, avoid over-engineering.

**Step 2 — Architecture:** Write `## Architecture` subsections first — Follow [Artifacts](references/solutioning-artifacts.md) (Step 3 reads inputs from these diagrams). Generate in order: (1) **System Landscape** — if PRD Requirements section (Iteration: also unincorporated `requirementsNN.md` Active sections) references 2+ distinct owned systems each with own deployment boundary, else skip; (2) **System Context** — if applicable per Artifacts conditions, else skip; (3) **Container Diagram** — always, immediately below Mermaid code block append: `**Owned containers:** {comma-separated list of containers team builds and deploys — not external systems, third-party services, or datastores}`; (4) **Deployment Diagram** — cloud-native (infrastructure.md Path B → Cloud-native sub-choice only): generate infrastructure-layer diagram; PaaS, Local-first (also Path B), Path A (managed platform): insert `<!-- Deployment Diagram: not applicable — {reason} -->`; (5) **Component Diagrams** — for each container in "Owned containers" list: generate `#### {Container Name}` subsection with Mermaid diagram showing internal components (controllers, services, repositories, handlers) and relationships, use module data from `specs/system/modules/*.md` if available else infer from Phase 2 flows and tech stack, if neither yields identifiable components insert `<!-- Component Diagram: not enough information to determine components for {Container Name} -->` and skip subsection; (6) **Code Diagrams** — only if `codebase_analysis` is not `skipped` AND at least one module has `scan_status: ok`: for each architecturally significant component (implementing cross-cutting concern, integration boundary, or core domain logic — typically 1–3 per container), generate `#### {Component Name}` subsection showing 2–3 key classes/modules and relationships, if condition not met omit `### Code Diagrams` grouping header entirely. Flag XL/XXL estimation risk. Embed unresolved questions inline within section they belong to — not as trailing `## Open Questions` section. For Iteration mode, `solution.md` is full, self-contained, updated architecture document (not delta) — regenerate all diagrams and sections to reflect complete architecture (prior and new elements together), keep "Owned containers" cumulative across iterations, incorporate new requirements and implementation changes from build history.

**Step 3 — Structure Planning:** Extract inputs per [references/architecture/code-structure.md](references/architecture/code-structure.md) § Inputs. Apply rules per [references/architecture/code-structure.md](references/architecture/code-structure.md) § Decision Rules (Rules 1–4), § Generation Rules (Rules 1–9), § Folder Tree Assembly to produce structure plan — record each decision with rationale. Output: hold structure plan in-memory for Step 4 writing (Code Structure section) — structure includes: repository_strategy decision + rationale, organization_pattern decision + rationale, monorepo_tool (if applicable) + rationale, layers list with purposes and dependencies, folder tree (complete structure with actual names), conventions (tests, configs, shared code, generated code), dependency rules (import constraints per layer).

**Step 4 — Write:** Generate remaining sections — Follow [Artifacts](references/solutioning-artifacts.md). **Code Structure:** Write `## Code Structure` (after `## Architecture`, before `## Tech Stack`) from Step 3 structure plan — Follow `## Code Structure` schema in [references/solutioning-artifacts.md](references/solutioning-artifacts.md). **Test framework:** Include in Tech Stack table and write to `system.md § Conventions / § Test Framework` (codebase-analysis does not populate this) — confirm runtime language first (Phase 3 resolves this). **Data schemas:** Define schemas for every data entity regardless of storage type — structure and field names must always be specified; types, constraints, contract details may use `{TBD: reason}` when implementation framework unconfirmed — add to Data Model section. If `codebase_analysis: skipped`, add at very top of `solution.md` (before any section header): "**Codebase analysis skipped** — this solution was drafted from PRD only. Architectural assumptions may not reflect the existing codebase." **Frontmatter:** Set `status: draft` and populate `reflects_requirements: []` — **Initial mode:** set to empty list `[]` (no requirements files incorporated yet, only prd.md); **Iteration mode:** append NN numbers from all newly incorporated `requirementsNN.md` files to existing list (e.g., if solution.md had `[01, 02]` and incorporating requirements03 and requirements04, update to `[01, 02, 03, 04]`). Compare output against `solution.md` schema in `references/solutioning-artifacts.md` — confirm all required keys present. After each section (Steps 2 and 4): record key decisions and trade-offs as rationale — if diverged from reference default, note why; if divergence applies broadly to other projects, update reference instead; when uncertain, record as project rationale. While writing (Steps 2 and 4), if identify cross-cutting dependency not in PRD dependency map, add to `## Technical Risks` with Risk = `**Discovered Prerequisite:** REQ-{ID} requires {dependency description} — not in PRD dependency map`; assess Likelihood/Impact/Mitigation. Do not modify `prd.md` — on gap or conflict: **absorb** if requirement can still be met (record as rationale note); **flag and stop** if infeasible or materially changes sizing (recommend re-running assessing).

**Step 5 — Sync solution.md to system.md:** Extract and write to system.md: (1) **§ Code Structure:** Copy from `solution.md § Code Structure` (already uses correct section name — no renaming needed); (2) **§ Foundation:** Append operational markers (Infra, Delivery, Environments) from `solution.md § Operational Concerns`; (3) **§ Architecture:** Append component list from "Owned containers" list in `solution.md § Architecture / ### Container Diagram`.

Proceed to Phase 5: Quality Review.

## Phase 5: Quality Review

Validate flow completeness and Code Structure completeness, spawn subagent review, present findings to user for apply/skip/discuss, and fix all applied findings before publishing.

**Step 1 — Pre-Review Validation:** **Flow completeness:** Validate flows per [references/solution-flows.md](references/solution-flows.md) — if issues found, **fix immediately** before proceeding to subagent review. **Code Structure completeness:** Follow [references/architecture/code-structure.md](references/architecture/code-structure.md) § Completeness Checklist — if issues found, **fix immediately** before proceeding to subagent review.

**Step 2 — Subagent Review:** Spawn subagent to review `solution.md` for issues — subagent is fresh, prompt must be self-contained. Follow [references/solution-review.md](references/solution-review.md) — fill in `{solution-path}` and `{requirements-path}` (path to PRD or requirements file loaded in Phase 1 Step 2). After subagent returns: **No findings:** note "Subagent review: no issues found" and proceed to Phase 6. **Findings present:** for each finding in order, present it (evidence, section, issue, recommendation, why) and ask "[1] apply  [2] skip  [3] discuss" — `apply` → update `solution.md` with recommendation, continue to next finding; `skip` → next finding; `discuss` → probe alternatives, if user confirms recommendation (original or revised) apply it, else treat as skip. When all findings processed, proceed to Phase 6.

Proceed to Phase 6: Publishing.

## Phase 6: Publishing

Set solution.md frontmatter to status final after completing all quality checks, and ensure Architecture subsections are in correct zoom-in order.

Finalizes solution design — set `solution.md` frontmatter to `status: final` after completing all quality checks; ensure `## Architecture` subsections in zoom-in order (System Landscape → System Context → Container → Deployment → Component → Code Diagrams) as specified in [Artifacts](references/solutioning-artifacts.md).

Proceed to Phase 7: Review.

## Phase 7: Review

Review mode or post-publish refinement - present artifact path, set to draft, conduct iterative revisions section-by-section, and finalize when complete.

Iterates on output with user until satisfied — present Markdown file path; set artifact to `status: draft` when beginning revisions; for each revision: update `solution.md`; ask "Anything else, or ready to move on?" and repeat until done (switch to section-by-section if asked); when review complete, set `status: final`.

Proceed to Phase 8: Retro.

## Phase 8: Retro

Reflect on the solutioning process and generate retro output for skill improvement.

Emits retro report covering full run — Follow [../references/neat-project/retro-template.md](../references/neat-project/retro-template.md) § Simple — `{skill-name}` = `neat-discovery-solutioning`.
