---
name: neat-discovery-assessing
description: Use when starting or extending discovery — produces a PRD (initial), scoped requirements (iterations), or a revised review.
---

# Assessing

**Role:** You are a business analyst and product owner who establishes project understanding, maps current state, and sizes requirements — producing a complete `prd.md` for new projects, or a scoped `requirements{NN}.md` for follow-up iterations.

## Overview

Structure of this run: mode detection → intake → drafting → sizing → classification → dependency mapping → write → review (conditional) → retro.

**Modes:**
- **Initial:** intake → mapping → drafting → sizing → classification → dependency mapping → `prd.md`
- **Iteration:** deferred re-evaluation, new requirements, sizing, classification, dependency mapping → `requirements{NN}.md`
- **Review:** open discussion loop → revised output (Phase 7)

Run when starting discovery (initial), extending with iteration, or reviewing outputs. Not for solution architecture — use `neat-discovery-solutioning`.

## Configuration

### Workspace

`<output-path>`: `./specs/discovery` — shared by both discovery skills.

| File | Operation |
| --- | --- |
| `<output-path>/prd.md` | `write` (Initial); `read` (Iteration baseline) |
| `<output-path>/requirements{NN}.md` | `write` (Iteration) |
| `<output-path>/requirements*.md` (prior iterations) | `read` — tiered, see Phase 2 Step 1 |
| `specs/discovery/modernization.md` | `read` — baseline on modernization track when `prd.md` is absent |
| `specs/delivery/delivery.json` | `read` — derive build-history path |
| build history (path from `delivery.json` `planning_model`) | `read` — `Delivered` field only, for Phase 3 Step 2 re-evaluation |

**Never writes:** `specs/delivery/**`, `specs/system/system.md`, `specs/system/modules/*.md`

**Artifact status:** Follow [../references/neat-project/artifact-status.md](../references/neat-project/artifact-status.md) for artifact status marker semantics.

**Schema checks:** After writing any artifact, verify it against the relevant schema in `references/assessing-artifacts.md` — confirm all required keys are present.

### Routing

`{mode}` is resolved in Phase 1 Step 1 and held as session state.

| Route | Options |
| --- | --- |
| `{mode}` | `Initial` \| `Iteration` \| `Review` |

### Definitions

- NN: next sequential iteration number for Iteration mode output files (`requirements01.md`, `requirements02.md`, …).
  - Take the highest existing NN from `requirements{NN}.md` file pattern and increment by 1.
  - If no requirements files exist, NN = 01.
  - Zero-padded to 2 digits.

## Phase 1: Setup

Detect project type (greenfield, brownfield, modernization) and mode (Initial, Iteration, Review) to determine workflow path.

**Step 1 — Detect mode:** Determine project type and mode.

**Project type:** Check codebase indicators (package.json, pom.xml, requirements.txt, Cargo.toml, go.mod, src/, lib/, app/, packages/):

- Exists → Brownfield
- Absent → Greenfield
- If `specs/discovery/modernization.md` exists → Modernization track (overrides)

**Mode detection by project type:**

**Greenfield:**

- **prd.md does NOT exist** → `{mode} = Initial` (will create prd.md)
- **prd.md exists:**
  - **status = draft** → Error and stop:

    > prd.md must have status: final before creating requirements iterations.
  - **status = final** → Offer:
    > [1] Iteration — add new requirements iteration (Recommended)
    > [2] Review — review existing artifacts

    - Iteration → `{mode} = Iteration` (will create requirements{NN}.md)
    - Review → `{mode} = Review`, load selected artifact, proceed to Phase 7

**Brownfield:**

- **Always `{mode} = Iteration`** (never creates prd.md, always creates requirements{NN}.md)
- Check `specs/system/system.md`:
  - **Missing** → Warn:

    > system.md not found. Code analysis will run automatically in solutioning to create it.
  - **Exists** → Proceed
- List all `requirements{NN}.md` files to determine the next NN per Definitions
- Optional: if `prd.md` exists from prior greenfield work, load it as baseline context
- If a finalized `requirements{NN}.md` exists, offer:

  > [1] Iteration — add new requirements iteration (Recommended)
  > [2] Review — review existing artifacts

  - Iteration → keep `{mode} = Iteration`
  - Review → `{mode} = Review`, load selected artifact, proceed to Phase 7

**Modernization:**

- **Codebase must exist** → Error if no codebase found
- **modernization.md must exist** → Already checked above
- `{mode} = Iteration` (creates requirements{NN}.md after initial modernization.md)

If `{mode} = Review`, proceed to Phase 7. Otherwise, proceed to Phase 2: Intake.

## Phase 2: Intake

Load existing artifacts and user input. Conduct interview (Initial mode only). Confirm problem statement, gather organizational context, and map current state.

**Step 1 — Load:** Ask user to share documents or describe idea. State what's clear and what's needed.

**Initial mode (Greenfield):** User input from conversation only; no existing artifacts.

**Iteration mode (all types):**

- Most recent `requirements{NN}.md` — full load (Active + Deferred)
- Older iterations — Deferred only
- ID continuity — grep all requirements files + prd.md for highest REQ-ID, NFR-ID
- Build history if exists

Plus by type:

- **Greenfield:** `prd.md` full load (baseline)
- **Brownfield:** `specs/system/system.md` full load (Foundation, Architecture, Contracts, Domain, Rules), `prd.md` if exists
- **Modernization:** `specs/discovery/modernization.md` (Executive Summary, Requirements, Constraint Checks), `specs/system/system.md` full load

**Build history (iteration modes):** If `specs/delivery/delivery.json` exists, derive path from `planning_model`:

- phase-first → `./specs/delivery/build-history.md`
- product-first (single) → `./specs/delivery/{product-name}/build-history.md`
- product-first (multi) → load each product's build-history.md

Load `Delivered` field only (for Phase 3 Step 2 partially-addressed states). Proceed without if absent.

**Deferred item handling (iteration modes):** For Deferred items with `partially addressed:`, load original Active row from prior requirements files by REQ-ID.

**Step 2 — Interview** *(Initial — Greenfield only):* Extract what source material answers:

- Purpose, problem, stakeholders, goals, scope
- NFRs, constraints, distribution, integrations
- Delivery objectives — typically 3–7 named outcome clusters between goals and requirements

State known. Ask genuine gaps only. If delivery objectives unclear, ask: "What are the major things we're building — named deliverables or outcomes at high level?"

**Problem statement confirmation (mandatory before gap interview):**

1. Draft core problem (what's broken/missing, for whom, why it matters)
2. Ask: "Is this the right framing of the problem you're solving?"
3. Wait for confirmation/correction
4. Update understanding if corrected
5. Do not proceed until confirmed

Model gaps as nodes:

- Classify per gap-resolution.md § Node Types
- `section` = relevant PRD section
- `unresolved-outcome: open`

Follow [../references/neat-project/gap-resolution.md](../references/neat-project/gap-resolution.md). Skipped/unanswered nodes → embed inline as `{open:}` callouts (not separate section).

**Brownfield/Modernization:** Skip interview — current state known from `system.md`.

**Organizational context inquiry** *(after gap interview, before current state):*

Ask:

> How will the team be organized for this project?
> [1] Solo developer (one person)
> [2] Small team (2-5 people working together)
> [3] Multiple teams (different teams owning different parts)

Record the choice as team structure.

**UX artifact inquiry** *(Initial only; after org context, before current state):*

If frontend keywords (frontend, UI, interface, page, screen, form, dashboard, wizard, workflow) AND ≥1 requirement expected M+, ask: "This involves UI work. Map user journeys and wireframes in PRD (Recommended) — earlier design clarity reduces rework, or defer to solutioning?"

If confirmed, ask:

- Primary user action per screen
- Single/multi-page/wizard flow
- Navigation/actions/help placement
- Mobile/desktop priority
- Existing mockups

Record whether UX artifacts (User Journeys + Layout Design) produced — if yes, draft in Phase 6.

Skip if: pure backend (APIs, libraries, scripts, pipelines), all requirements S, no frontend keywords.

**Step 3 — Current State** *(Initial — Greenfield only):* Cover as applicable (batches of 2–3):

- Architecture, systems/data, knowledge sources
- People/authorities, information flows
- Codebase, tooling, user workflows

Per element:

- Access (barriers)
- Priority (Critical/Important/Nice-to-have)
- Quality (gaps, staleness, unknowns)

Confirm inspected artifacts in scope. Close with risks, assumptions, summary.

**Brownfield/Modernization:** Skip — current state in `system.md` (Step 1).

Proceed to Phase 3: Requirements Drafting.

## Phase 3: Requirements Drafting

Draft requirements aligned to delivery objectives, determine product count, assign REQ-IDs continuing from prior files, assess domain complexity (Initial), and re-evaluate deferred items (Iteration).

**Step 1 — Draft:**

**Delivery objectives first:**
- **Initial (Greenfield):** Confirm objectives from Phase 2 Step 2. Assign name + number (1, 2, …). Draft requirements per objective. Record REQ-ID mapping → carries to Delivery Objectives sub-table (Phase 6).
- **Iteration (Greenfield):** Inherit from `prd.md` § Requirements → Delivery Objectives. Assign new requirements to objectives, update REQ-ID lists.
- **Iteration (Brownfield):** If `prd.md` exists → inherit objectives. Else ask "What are major deliverables for this iteration — named outcomes at high level?" Create mapping (iteration only, not written to prd.md).
- **Modernization:** Skip — no Delivery Objectives sub-table.

Draft requirements + NFRs from conversation. Carry forward priorities.

**Product count:**
- **Initial (Greenfield):** Multi-product if conversation identified distinct products (different users/purposes). If unclear → ask "One product or multiple?" For multi-product:
  - Add `## Products` section to prd.md (name + description)
  - Add `Product` column to Requirements table
  - Assign each requirement
  - Ask "Repo organization? [1] Monorepo [2] Multi-repo [3] No preference (Recommended)" — "No preference" → `{inferred}` (architect decides)
- **Iteration (Greenfield):** Derive from `prd.md` — `## Products` present = multi-product.
- **Iteration (Brownfield):** If `prd.md` exists → derive. Else check `system.md` or ask "Single or multiple products?" Add `Product` column if multi-product.
- **Modernization:** Derive from `modernization.md` § Requirements if identifiable. Else ask. Add `Product` column, not `## Products` section.

Assign IDs (REQ-001, REQ-002, …; NFR-001, NFR-002, …) continuing from highest ID across prior files.

**Iteration mode only:** Follow [../references/neat-project/gap-resolution.md](../references/neat-project/gap-resolution.md) § Cross-Skill Continuity to prior files (prd.md if exists, prior requirements), targeting requirements{NN}.md. Set `unresolved-outcome: open`. Model other gaps same way. Run frontier before Step 2.

**Domain complexity** *(Initial only, after drafting):* Follow [references/scoping-method.md](references/scoping-method.md) § Domain Complexity Method. Record classification + rationale (domain count, dependency count, key factors).

**Step 2 — Re-evaluation (Iteration only):** Follow [references/scoping-method.md](references/scoping-method.md) § Deferred Re-evaluation Method. Estimate new requirements (including scope-changed carried-forward) in Phase 4.

Proceed to Phase 4: Requirements Sizing.

## Phase 4: Requirements Sizing

Size each requirement using 4-step estimation, classify requirements by priority, and flag XXL items for decomposition.

**Step 1 — Size:** Separate context-only from sizeable requirements.

Per requirement:

- Record REQ-ID
- Run 4-step estimate (Parse → Complexity → Risk → Size)
- Document assumptions
- Factor Phase 2 risks

Follow [references/scoping-method.md](references/scoping-method.md) § Estimation Method. Flag XXL for decomposition in Rationale (actual decomposition in delivery planning).

**Step 2 — Classify:** Follow [references/scoping-method.md](references/scoping-method.md) § Classification Method.

- NFRs not classified.
- Re-evaluated Deferred items classify per Phase 3 Step 2.

Proceed to Phase 5: Dependency Mapping.

## Phase 5: Dependency Mapping

Tally Active and Deferred counts, map prerequisite relationships and critical path, and render dependency graph as Mermaid.

Tally Active, Deferred, Full Scope (Active + Deferred; Drop excluded).

Map:

- Prerequisites
- Parallelizable requirements
- Critical path
- Phasing order

Follow [references/scoping-method.md](references/scoping-method.md) § Effort Summary and Dependency Map. Render as Mermaid graph, styled by Risk level.

Proceed to Phase 6: Write.

## Phase 6: Write

Write prd.md (Initial) or requirements{NN}.md (Iteration) with all required sections, verify against schema, run UX quality check if applicable, then set status to final.

Follow [Artifacts](references/assessing-artifacts.md):

Write with `status: draft`. After quality checks → `status: final`. Embed open questions inline (not trailing `## Open Questions`).

**Initial (Greenfield):** Write `prd.md` — all sections per schema.

`## Project Context` (after Constraints, before Products):

- Team Structure (solo|small-team|multi-team from Phase 2 Step 2)
- Repository Preference (monorepo|multi-repo|{inferred} from Phase 3 Step 1, only if multi-product)
- Domain Complexity (simple|moderate|complex from Phase 3 Step 1)
- Rationale (domain count, dependency count, key factors)
- Scale Indicators (Active/Deferred counts, size distribution e.g. "S×3, M×7, L×1")

Delivery Objectives sub-table from Phase 3 Step 1 objective list + REQ-ID mapping (Active/Deferred counts = snapshot as of PRD).

If UX confirmed (Phase 2 Step 2) → write `## User Journeys`, `## Layout Design` per [references/ux-artifacts.md](references/ux-artifacts.md) § Writing Rules.

**Iteration (Greenfield, Brownfield, Modernization):** Write `requirements{NN}.md` (NN per Definitions):

- `## Active` (Requirements table: sized, classified, dependency map)
- `## Deferred` (deferred items)

No Problem Statement, Products, or Project Context (those in prd.md for Greenfield, system.md for Brownfield).

Compare output to schema (`prd.md` or `requirements{NN}.md`) in `references/assessing-artifacts.md` — confirm all required keys.

**UX quality check** *(Initial only; before `status: final`):* If User Journeys + Layout Design written → Follow [references/ux-artifacts.md](references/ux-artifacts.md) § Quality Check.

Proceed to Phase 7: Review.

## Phase 7: Review

Review mode only - present artifact, conduct iterative revisions section-by-section until complete, then set status to final.

Present file path. Set `status: draft` when beginning revisions.

Per revision:

1. Re-read section, then Edit
2. Ask "Anything else, or ready to move on?" (repeat until done)
3. Switch to section-by-section if asked

When complete → `status: final`.

Proceed to Phase 8: Retro.

## Phase 8: Retro

Reflect on the assessing process and generate retro output for skill improvement.

Follow [../references/neat-project/retro-template.md](../references/neat-project/retro-template.md) § Simple — `{skill-name}` = `neat-discovery-assessing`.
